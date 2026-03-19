# Workflow Specification Reference

Complete workflow.yaml schema, trigger types, config block, and validation rules.

## workflow.yaml Schema Quick Reference

All top-level fields:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Workflow identifier |
| `version` | Yes | Semantic version string (e.g., `"1.0"`) |
| `description` | No | Human-readable description |
| `metadata` | No | Discovery metadata: `owner`, `team`, `environment`, `tags` |
| `triggers` | Yes | List of trigger definitions (at least one) |
| `contract` | Yes | Dependency contract (version + dependencies map) |
| `nodes` | Yes | Map of node names to node definitions (each with `path:`) |
| `edges` | Yes | Top-level list of `{from: node, to: node}` pairs |
| `config` | No | Open config block: `timeout`, `retries`, and any custom keys |

Critical schema rules:
- Nodes use `path:` (not `source:`). `source:` is rejected at validate time.
- Edges are a separate top-level `edges:` list. Do not use `depends_on:`
  inside node blocks -- it is not a recognized field and is silently ignored.

## Minimal Example

```yaml
name: my-workflow
version: "1.0"
description: "A minimal workflow"

# Optional: metadata for MCP discovery (wf_list / wf_describe)
# metadata:
#   owner: my-team
#   tags: [dev]

triggers:
  - type: manual

contract:
  version: "1"
  dependencies: {}

nodes:
  hello:
    path: ./nodes/hello.ts

edges: []

config:
  timeout: 30s
  retries: 0
```

## Trigger Types

| Type | Mechanism | Required Fields | K8s Resources | Status |
|------|-----------|----------------|---------------|--------|
| `manual` | MCP server POSTs to `/run` via direct HTTP | none | -- | Implemented |
| `cron` | MCP server internal scheduler reads annotation, calls wf_run | `schedule`, optional `name` | Deployment annotation | Implemented |
| `queue` | NATS subscription -> execute | `subject` | -- | Implemented |
| `webhook` | Future: gateway -> NATS bridge | `path` | -- | Roadmap |

### Trigger Name Field

Triggers can have an optional `name` field (must match `[a-z][a-z0-9_-]*`,
unique within workflow). Named cron triggers POST `{"trigger": "<name>"}` to
`/run`. Root nodes receive this as `input.trigger` to branch behavior.

## Cron Trigger Lifecycle

1. Define in workflow.yaml: `type: cron`, `schedule: "0 9 * * *"`,
   optional `name`.
2. `tntc deploy` records the schedule in a `tentacular.dev/cron-schedule`
   annotation on the Deployment. No CronJob resource is created.
3. The MCP server's internal cron scheduler reads the annotation on startup
   and after each `wf_apply`. It fires `wf_run` internally on schedule.
4. Named triggers: scheduler POSTs `{"trigger": "<name>"}` to `/run`.
5. `tntc undeploy` removes the Deployment (and its annotation). The MCP
   server drops the cron entry automatically on the next sync.

## Module Pre-Warm and First-Start Race

When deploying workflows with `jsr:` or `npm:` dependencies for the first
time, the MCP server pre-warms the module proxy cache in the background
immediately after `wf_apply` returns. There is a brief race window where the
workflow pod may start before warming completes. If this happens, the first
pod start will fail with a module resolution timeout, but Kubernetes will
automatically restart the pod. By the second attempt, the cache will be warm
and the pod will start successfully.

This is expected and normal for first-time deploys with new module
dependencies. Confirm recovery by checking `wf_pods` for restart count and
`wf_logs` for the error on the first attempt.

## Queue Trigger (NATS)

Queue triggers subscribe to NATS subjects. Config: `config.nats_url`, auth:
`secrets.nats.token`. If either is missing, the engine warns and skips NATS
(graceful degradation). Messages are parsed as JSON and passed as input to
root nodes.

## Config Block

The `config:` block is open -- it accepts arbitrary keys alongside `timeout`
and `retries`. Custom keys flow through to `ctx.config` in nodes. This is
the standard mechanism for non-secret workflow configuration.

```yaml
config:
  timeout: 30s
  retries: 2
  nats_url: "nats.example.com:4222"
  custom_setting: "value"
```

In Go, extra keys are stored in `WorkflowConfig.Extras` (via
`yaml:",inline"`). Use `ToMap()` to get a flat merged map.

## Schema Gotchas

- Use `path:` in node definitions, never `source:`.
- Use top-level `edges:` list, never `depends_on:` inside a node block.
- `edges:` must be present even if empty (`edges: []`).
- `contract.version` is a string (`"1"`), not an integer.
