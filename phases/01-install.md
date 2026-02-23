# Phase 01: Install tntc

**Gate: `tntc version` must succeed AND the engine directory must be present before any other work.**

## Check

```bash
which tntc && tntc version
ls ~/.tentacular/engine/main.ts 2>/dev/null && echo "engine ok" || echo "engine MISSING"
```

If both succeed, you are done with this phase.

## Install tntc binary

If `tntc` is missing:

```bash
curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/install.sh | sh
```

## Install the engine

The install script only downloads the `tntc` binary. `tntc test` and `tntc dev` also
require the Deno engine directory at `~/.tentacular/engine/`. Install it manually:

```bash
git clone --depth=1 https://github.com/randybias/tentacular /tmp/tentacular-src
mkdir -p ~/.tentacular
cp -r /tmp/tentacular-src/engine ~/.tentacular/engine
rm -rf /tmp/tentacular-src
```

Verify:
```bash
ls ~/.tentacular/engine/main.ts   # must exist
```

⚠️ This is a known gap in the install script. Until the release bundles the engine,
this manual step is required after every clean install.

## Cluster prerequisites (first-time setup only)

If this is a new cluster, gVisor must be installed before any workflow deploy will work.
`tntc cluster check` will catch this, but you need the fix commands:

```bash
# 1. On each cluster node (requires root/SSH access):
sudo bash <(curl -fsSL https://raw.githubusercontent.com/randybias/tentacular/main/deploy/gvisor/install.sh)

# 2. Create the RuntimeClass (run once per cluster):
kubectl apply -f https://raw.githubusercontent.com/randybias/tentacular/main/deploy/gvisor/runtimeclass.yaml

# 3. Verify:
kubectl apply -f https://raw.githubusercontent.com/randybias/tentacular/main/deploy/gvisor/test-pod.yaml
kubectl wait --for=condition=Ready pod/gvisor-test --timeout=60s
kubectl logs gvisor-test   # should show gVisor kernel messages
kubectl delete pod gvisor-test
```

Skip this block if the cluster already has gVisor (profile will show `Use runtime_class: gvisor`).
If gVisor is intentionally absent (e.g., kind cluster), set `runtime_class: ""` in config.

## Verify

| Output | Meaning | Action |
|--------|---------|--------|
| `tntc v1.x.x` + `engine ok` | Ready | Proceed to phase 02 |
| `dev (commit none, built unknown)` + `engine ok` | Source build | Proceed to phase 02 |
| `tntc version` fails | Install failed | Stop. Surface error. Do not proceed. |
| `engine MISSING` | Engine not installed | Run the engine install block above |

**If either check fails: stop completely.** Do not attempt any other tntc commands.

## Gate Verification

Before leaving this phase, confirm:

- [ ] `tntc version` exits 0
- [ ] `~/.tentacular/engine/main.ts` exists
- [ ] gVisor is installed on target cluster (or `runtime_class: ""` is planned for config)
