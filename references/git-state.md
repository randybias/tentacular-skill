# Git-Backed State Reference

Git-backed state is an optional feature that makes a git monorepo the system
of record for all tentacle source code, metadata, and optionally encrypted
secrets. When enabled, agents commit to git before deploying — git becomes
the authoritative source from which the cluster can be fully reconstructed.

## When It Is Enabled

Git-backed state is enabled when IT provides a git URL and credentials in
the tentacular config. Without these, tentacle source lives on disk only
(`~/tentacles/<enclave>/<tentacle>/`).

Check with `tntc state status` to see whether git-state is configured and
whether the working tree is clean.

## The Three-Layer Mental Model

Every tentacle exists in three layers simultaneously:

```
Git (system of record)
    enclaves/<enclave>/<tentacle>/workflow.yaml
    enclaves/<enclave>/<tentacle>/nodes/*.ts
    enclaves/<enclave>/<tentacle>/CONTEXT.md
           │
           ▼
Disk (working copy)
    ~/tentacles/<enclave>/<tentacle>/workflow.yaml
    ~/tentacles/<enclave>/<tentacle>/nodes/*.ts
           │
           ▼
K8s (deployed runtime)
    namespace: <enclave>
    deployment: <tentacle>
    configmap:  tentacle source code
```

All three must agree on the same version before a deployment is considered
complete.

## Artifact Mapping

| Artifact | Git Path | Disk Path | K8s Resource |
|----------|----------|-----------|--------------|
| Workflow spec | `enclaves/<enclave>/<tentacle>/workflow.yaml` | `~/tentacles/<enclave>/<tentacle>/workflow.yaml` | ConfigMap `<tentacle>-source` |
| Node code | `enclaves/<enclave>/<tentacle>/nodes/*.ts` | `~/tentacles/<enclave>/<tentacle>/nodes/*.ts` | ConfigMap `<tentacle>-source` |
| Parameter schema | `enclaves/<enclave>/<tentacle>/params.schema.yaml` | `~/tentacles/<enclave>/<tentacle>/params.schema.yaml` | Not deployed |
| Design intent | `enclaves/<enclave>/<tentacle>/CONTEXT.md` | `~/tentacles/<enclave>/<tentacle>/CONTEXT.md` | Not deployed |
| Tentacle metadata | `enclaves/<enclave>/<tentacle>/tentacle.yaml` | (snapshot only) | K8s Deployment annotations |
| Enclave metadata | `enclaves/<enclave>/enclave.yaml` | (snapshot only) | K8s namespace annotations |
| User secrets | `enclaves/<enclave>/.secrets/<tentacle>.enc.yaml` | `~/tentacles/<enclave>/<tentacle>/.secrets.yaml` | K8s Secret |

## Deploy Flow (git-state enabled)

When git-state is enabled, the deploy flow gains a commit-and-verify gate
between validation and deployment:

```
1. tntc validate                         # validate spec + contract
2. tntc visualize --rich --write         # persist contract artifacts
3. tntc test -o json                     # mock tests
4. tntc test --live --cluster <target>   # live tests
   --- git-state gate ---
5. Write CONTEXT.md (if new or changed)
6. tntc state commit "deploy(<enclave>/<tentacle>): <message>"
7. tntc state status --assert-clean      # refuse if dirty
   --- deploy ---
8. tntc deploy -o json
9. tntc run <name> -o json               # post-deploy verification
```

Steps 5-7 are the deploy gate. `tntc deploy` checks for a clean git tree
and refuses to proceed if uncommitted changes exist in the enclave/tentacle
path. This ensures every deployed state has a corresponding git commit.

## CONTEXT.md

`CONTEXT.md` is the most important file for cross-agent continuity. Write
it when creating a new tentacle and update it when making significant changes.

```markdown
# <tentacle-name>

## Original Request

<What the user asked for, in their words or a faithful paraphrase>

## Design Decisions

<Key choices made during development and why. Examples:>
- Chose headless browser over simple HTTP fetch because competitor sites
  render prices client-side
- Cron schedule set to 6am UTC per user request (before US market open)

## Known Limitations

<What this tentacle does NOT handle, and why>

## Iteration Notes

<Significant changes across versions, beyond what git log shows>
```

Always write `CONTEXT.md` before the first deploy. Update it when:
- The tentacle's purpose changes
- A significant design decision is made during an iteration
- A known limitation is discovered

## Commit Message Convention

```
deploy(<enclave>/<tentacle>): <meaningful message>
```

Examples:
```
deploy(competitor-pricing/price-monitor): initial deploy — basic price scraper for 5 sites
deploy(competitor-pricing/price-monitor): add headless browser sidecar for JS-heavy sites
deploy(infra-alerts/node-health): increase cron frequency from hourly to 15-min
```

The commit message is the primary record of deploy intent. Write it as a
human-readable summary of what changed and why, not just "update".

## Deploy Gate: Refusing a Dirty Tree

If `tntc state status` reports uncommitted changes in the enclave/tentacle
path, do NOT proceed with deployment. The deploy gate exists to ensure that
every deployed state is recoverable from git.

Common causes:
- `CONTEXT.md` not created or updated before deploying a new tentacle
- Node code edited locally but not staged and committed
- `workflow.yaml` updated but not committed

Fix: stage all changes in the tentacle directory, write or update
`CONTEXT.md`, then run `tntc state commit "<message>"`.

## Archive Flow

When a tentacle is decommissioned:

```bash
# 1. Undeploy from cluster
tntc undeploy <tentacle> --enclave <enclave>

# 2. Move in git
git -C <state-repo> mv enclaves/<enclave>/<tentacle>/ archive/<enclave>/<tentacle>/

# 3. Write ARCHIVE.md
# (see template below)

# 4. Commit
git -C <state-repo> add -A
git -C <state-repo> commit -m "archive(<enclave>/<tentacle>): <reason>"
```

`ARCHIVE.md` template:

```markdown
# Archive Record

**Archived:** <RFC3339 timestamp>
**Archived by:** <email>
**Reason:** <why the tentacle was retired>
**Last deployed version:** <version from workflow.yaml>
**Final state:** <brief description of operational state at retirement>
```

The git history preserves the full lifecycle — `git log archive/<enclave>/<tentacle>/`
shows every commit from initial creation to retirement.

## tntc state Commands

| Command | Description |
|---------|-------------|
| `tntc state init` | Initialize git-state repo configuration for this tentacular workspace |
| `tntc state status` | Show working tree status relative to HEAD for all enclave paths |
| `tntc state commit "<message>"` | Stage all changes in the current enclave/tentacle directory and commit |
| `tntc state status --assert-clean` | Exit non-zero if working tree is dirty (used in deploy gate) |

## Common Mistakes

| Mistake | What Happens | Fix |
|---------|--------------|-----|
| Deploying without committing CONTEXT.md | Deploy gate blocks | Write CONTEXT.md, run `tntc state commit` |
| Committing with generic message "update workflow" | History is unreadable | Use `deploy(<enclave>/<tentacle>): <meaningful message>` |
| Editing node code after the commit but before `tntc deploy` | Working tree becomes dirty, gate blocks | Re-commit before deploying |
| Deleting a tentacle directory on disk without archiving | State repo diverges from cluster reality | Always use `tntc undeploy` + `git mv` + ARCHIVE.md |
| Running `tntc state commit` from the wrong directory | Wrong files staged | Always run from `~/tentacles/<enclave>/<tentacle>/` or pass the path explicitly |
