# Scaffold Lifecycle Reference

## Lifecycle Overview

All paths from idea to deployed tentacle:

```
                    Scaffold (private or public)
                         |
                         | tntc scaffold init <scaffold> <name>
                         v
Fresh start -------> Tentacle (local)
  tntc init <name>       |
                         | configure, modify, extend, test
                         | tntc validate, tntc dev
                         v
                    Tentacle (local, validated)
                         |
                         | tntc deploy
                         v
                    Tentacle (deployed, on cluster)
```

Reverse path (tentacle to scaffold):

```
Tentacle (local)
    |
    | tntc scaffold extract
    v
Private scaffold (~/.tentacular/scaffolds/)
    |
    | (optional) Human review, PR to tentacular-scaffolds
    v
Public quickstart (tentacular-scaffolds/quickstarts/)
```

## The Four Lifecycles

**A. From scratch** -- No scaffold matches. Agent creates everything via
`tntc init`. No `params.schema.yaml` involved.

**B. Scaffold used as-is** -- Direct match found (private or public). Agent
fills in parameter values using the Parameter Negotiation Protocol, deploys.

**C. Scaffold as starting point, heavily modified** -- Close match found.
Agent uses the scaffold's structure and code as a base, then restructures
freely using the Scaffold Modification Protocol.

**D. Tentacle extracted to new scaffold** -- A working tentacle is extracted
as a reusable scaffold using the Scaffold Extraction Protocol (private by
default, optionally contributed to public quickstarts).

---

## Workspace Layout

```
~/tentacles/                            # ALL tentacles live here (flat)
+-- acme-uptime/                        #   a tentacle
+-- regional-monitor/                   #   a tentacle
+-- api-latency-reporter/               #   a tentacle

~/.tentacular/                          # SYSTEM directory
+-- config.yaml                         #   CLI configuration
+-- cache/
|   +-- scaffolds-index.yaml            #   cached quickstart index
+-- scaffolds/                          #   PRIVATE scaffolds
|   +-- our-standard-monitor/
|   |   +-- scaffold.yaml
|   |   +-- workflow.yaml
|   |   +-- params.schema.yaml
|   |   +-- nodes/
|   +-- our-etl-pipeline/
+-- quickstarts/                        #   LOCAL CACHE of public quickstarts
    +-- uptime-tracker/
    +-- site-change-detector/
```

Key rules:
- `~/tentacles/` is flat. Every directory is a tentacle, no exceptions.
- Private scaffolds (`~/.tentacular/scaffolds/`) are searched first.
- Public quickstarts (`~/.tentacular/quickstarts/`) are a local cache of the
  `tentacular-scaffolds` repo, refreshed by `tntc scaffold sync`.
- `~/.tentacular/` and its subdirectories are created with 0700 permissions
  (owner-only access). Private scaffolds may contain org-specific patterns,
  internal URLs, or architectural details that should not be world-readable.

### Name Validation

Scaffold and tentacle names must be valid kebab-case identifiers:
- Pattern: `^[a-z0-9][a-z0-9-]*[a-z0-9]$`
- Maximum length: 64 characters
- No path separators (`/`, `\`), no `..`, no special characters
- Examples: `uptime-tracker`, `our-etl-pipeline`, `e2e-exoskeleton-test`

This validation is enforced by `tntc scaffold init`, `tntc scaffold extract`,
and `tntc init`. It prevents path traversal attacks when names are used to
construct file paths.

### Scaffold Search Order

1. **Private scaffolds** (`~/.tentacular/scaffolds/`) -- org-specific, most
   likely to match
2. **Public quickstarts** (`~/.tentacular/quickstarts/`) -- community scaffolds
3. **Build from scratch** -- no scaffold matches, agent creates everything

---

## `tntc scaffold` Command Reference

### `tntc scaffold list`

List available scaffolds from private and public sources.

```
tntc scaffold list [flags]

Flags:
  --source <private|public|all>   Filter by source (default: all)
  --category <category>           Filter by category
  --tag <tag>                     Filter by tag
  --json                          Output as JSON
```

### `tntc scaffold search <query>`

Full-text search across scaffold name, displayName, description, and tags.

```
tntc scaffold search <query> [flags]

Flags:
  --source <private|public|all>   Filter by source (default: all)
  --json                          Output as JSON
```

### `tntc scaffold info <name>`

Show scaffold metadata, parameter summary, and file list.

```
tntc scaffold info <name> [flags]

Flags:
  --source <private|public>    Disambiguate if same name in both sources
```

### `tntc scaffold init <scaffold> <name>`

Create a tentacle from a scaffold.

```
tntc scaffold init <scaffold-name> <tentacle-name> [flags]

Flags:
  --source <private|public>    Disambiguate scaffold source (default: auto)
  --params-file <path>         Apply parameter values from YAML file
  --no-params                  Copy scaffold as-is without applying parameters
  --namespace <ns>             Set deployment.namespace in workflow.yaml
  --dir <path>                 Override output directory (default: ~/tentacles/<name>/)
```

Behavior:
1. Resolve scaffold (private first, then public, unless `--source` specified)
2. Copy scaffold files to `~/tentacles/<tentacle-name>/`
3. Create `tentacle.yaml` with provenance metadata
4. With `--params-file`: validate and apply parameter values to workflow.yaml
5. With `--no-params`: copy as-is, write `params.yaml.example`
6. Default (no flag): print parameter summary, write `params.yaml.example`

### `tntc scaffold extract`

Create a scaffold from a working tentacle. Run from within a tentacle
directory.

```
tntc scaffold extract [flags]

Flags:
  --name <name>         Name for the scaffold (default: derived from tentacle)
  --private             Save to ~/.tentacular/scaffolds/<name>/ (default)
  --public              Save to ./scaffold-output/ for PR to tentacular-scaffolds
  --output <path>       Override output directory
  --json                Output analysis as JSON without generating files
```

### `tntc scaffold sync`

Refresh local cache of public quickstarts from the remote repo.

```
tntc scaffold sync [flags]

Flags:
  --force    Bypass cache TTL and force refresh
```

### `tntc scaffold params show`

Show current parameter values for a tentacle. Run from within a tentacle
directory. Reads `params.schema.yaml` and resolves each path expression
against the current `workflow.yaml`.

### `tntc scaffold params validate`

Check that parameter values in `workflow.yaml` differ from the scaffold's
example values. Warns if any parameter still matches the example (likely not
configured yet). Run from within a tentacle directory.

### `tntc init <name>` (top-level, not under scaffold)

Create an empty tentacle from scratch. This is Lifecycle A -- no scaffold
involved. Creates `tentacle.yaml` with name and timestamp, skeleton
`workflow.yaml`, `nodes/hello.ts`, test fixtures, `.secrets.yaml.example`,
and `.gitignore`.

---

## Artifact Files

### scaffold.yaml

Metadata for a scaffold. Lives in scaffold directories.

```yaml
name: uptime-tracker
displayName: "Uptime Tracker"
description: "Probe HTTP endpoints every 5 minutes..."
category: monitoring
tags: [uptime-monitoring, time-series, postgres-state]
author: randybias
version: "1.0"
minTentacularVersion: "0.5.0"
complexity: moderate
```

### tentacle.yaml

Identity and provenance for a local tentacle. Created by `tntc init` or
`tntc scaffold init`.

From a scaffold:
```yaml
name: acme-uptime
created: 2026-03-22T14:30:00Z
scaffold:
  name: uptime-tracker
  version: "1.0"
  source: public
  modified: false
```

From scratch:
```yaml
name: custom-monitor
created: 2026-03-22T15:00:00Z
```

### params.schema.yaml

Declares user-configurable parameters. Tells agents what questions to ask
when instantiating a scaffold.

```yaml
version: "1"
description: "Parameters for the uptime-tracker scaffold"

parameters:
  endpoints:
    path: config.endpoints
    type: list
    items: map
    description: "HTTP endpoints to probe for uptime monitoring."
    required: true
    example:
      - url: "https://example.com"
        expected_body: ""

  latency_threshold_ms:
    path: config.latency_threshold_ms
    type: number
    description: "Maximum acceptable response time in ms before alerting"
    required: false
    default: 2000

  probe_schedule:
    path: triggers[name=check-endpoints].schedule
    type: string
    description: "Cron expression for probe frequency"
    required: false
    default: "*/5 * * * *"
```

**Parameter fields:** `path` (required, path expression into workflow.yaml),
`type` (required: string|number|boolean|list|map), `description` (required),
`required` (required: boolean), `default` (optional), `example` (optional),
`items` (optional, for list type).

### Path Expression Syntax

Path expressions in `params.schema.yaml` point to locations in
`workflow.yaml`. Two segment types:

| Segment | Syntax | Example |
|---------|--------|---------|
| Key | `<name>` | `config` |
| Filtered | `<name>[<field>=<value>]` | `triggers[name=check-endpoints]` |

Full path: segments joined by `.` (dot).

Examples:
- `config.endpoints` -- config -> endpoints
- `config.latency_threshold_ms` -- config -> latency_threshold_ms
- `triggers[name=check-endpoints].schedule` -- triggers -> find entry where
  name=check-endpoints -> schedule

---

## Parameter Negotiation Protocol (Lifecycle B)

When using a scaffold as-is:

1. `tntc scaffold init <scaffold> <name> --no-params`
2. Read `params.schema.yaml` from the tentacle directory
3. For each `required: true` parameter:
   - Present name, description, type, and example to the human
   - Collect the value
4. For each `required: false` parameter:
   - Present with its default value
   - Ask if the human wants to change it
5. Edit `workflow.yaml` directly at the paths indicated by the schema
6. `tntc scaffold params validate` -- verify no example values remain
7. `tntc validate`

**Alternative:** Write a `params.yaml` with all values upfront, use
`tntc scaffold init <scaffold> <name> --params-file params.yaml`.

---

## Scaffold Modification Protocol (Lifecycle C)

When using a scaffold as a starting point with structural changes:

1. `tntc scaffold init <scaffold> <name> --no-params`
2. Read the scaffold's `workflow.yaml` and node code
3. Present a modification plan to the human:
   - What changes (config structure, nodes, contract)
   - What stays (reusable logic, patterns)
4. After human approval, make all changes to tentacle files
5. Delete or rewrite `params.schema.yaml`:
   - Delete if structure diverged significantly
   - Rewrite if you anticipate future extraction
6. Set `modified: true` in `tentacle.yaml` scaffold section
7. `tntc validate`

---

## Scaffold Extraction Protocol (Lifecycle D)

When extracting a scaffold from a working tentacle:

1. `tntc scaffold extract --json` -- analysis without file generation
2. Review proposed parameters against extraction heuristics (below)
3. Present parameterization plan to the human
4. After approval, `tntc scaffold extract` to generate files
5. Default: private scaffold at `~/.tentacular/scaffolds/<name>/`
6. To publish: `--public`, review files, PR to `tentacular-scaffolds`

---

## Extraction Heuristics

### Parameterize (replace with safe examples)

| Signal | Reason |
|--------|--------|
| URL with a custom domain (not a well-known API) | Org-specific |
| Organization/team/project names | Org-specific |
| S3/storage bucket names (other than `tentacular`) | Org-specific |
| Slack channel names or webhook context | Org-specific |
| GitHub org/repo references | Org-specific |
| Email addresses or notification targets | Org-specific |
| Custom hostnames or internal DNS names | Org-specific |

### Parameterize with current value as default

| Signal | Reason |
|--------|--------|
| Numeric thresholds (latency, retry count, timeout) | Tunable defaults |
| Cron schedules | Tunable defaults |
| Batch sizes, concurrency limits | Structural but tunable |

### Do NOT parameterize

| Signal | Reason |
|--------|--------|
| Well-known API hosts (`api.anthropic.com`, `hooks.slack.com`) | Fixed by provider |
| Protocol and port for well-known services | Fixed |
| Contract dependency structure | Structural |
| Exoskeleton dependency names (`tentacular-*`) | Structural -- prefix is a security boundary (see below) |
| Exoskeleton dependency protocols | Structural -- fixed by service type |
| Node graph structure (edges, node list) | Structural -- the point of the scaffold |

### The `tentacular-` Prefix as a Security Boundary

The `tentacular-` prefix on dependency names is not just a naming convention --
it is the mechanism that triggers automatic exoskeleton provisioning. The MCP
server recognizes this prefix and provisions backing services (databases, object
storage, message queues), injects credentials, and fills in connection details.

Security implications:
- **Never parameterize `tentacular-*` dependency names.** Changing the name
  breaks auto-provisioning. The prefix is structural, not configurable.
- **Only three known service names exist:** `tentacular-postgres`,
  `tentacular-rustfs`, `tentacular-nats`. Any other `tentacular-*` name will
  not be recognized by the exoskeleton.
- **Deploying a scaffold means trusting its contract.** Any scaffold that
  declares a `tentacular-*` dependency will trigger provisioning of that
  service when deployed. Review contract declarations before deploying
  scaffolds from untrusted sources.
- **Exo-managed deps declare only `protocol`.** Do not add `host`, `port`,
  `auth`, or `database` fields -- the MCP server fills these at deploy time.

---

## Post-Deployment Prompt

After a successful `tntc deploy`, if the tentacle was created from a modified
scaffold (Lifecycle C) or from scratch (Lifecycle A), ask the user:

> "This tentacle is working well. Would you like me to extract it as a
> reusable scaffold for future use?"

This grows the scaffold library organically from real work.
