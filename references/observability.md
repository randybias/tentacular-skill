# Observability Reference

Tentacular uses OpenTelemetry (OTel) for distributed tracing, metrics, and
logs. When the observability stack is deployed, workflow pods emit telemetry
automatically -- no code changes required. LLM API calls are enriched with
token usage attributes via a GenAI fetch wrapper.

## Detecting Observability Availability

Check `cluster_profile` output. When the observability stack is installed, the
profile includes an `Observability` field:

```
Observability:
  available: true
  collector: otel-collector.tentacular-observability.svc.cluster.local:4318
```

If `Observability` is absent or `available: false`, the stack is not installed.
OTel env vars are still injected into pods but telemetry exports silently fail
with no impact on workflow execution.

## What Happens Automatically

When a workflow pod starts with `OTEL_DENO=true` (injected by the builder),
Deno's native OTel integration provides zero-code instrumentation:

| Signal | What Is Captured |
|--------|------------------|
| Traces | `Deno.serve` incoming request spans |
| Traces | `fetch()` outgoing request spans (LLM calls, sidecar calls, external APIs) |
| Traces | W3C Trace Context propagation across services |
| Logs | `console.*` calls become OTel log records, correlated with the active span |
| Metrics | HTTP server request duration, active requests, body sizes |

The engine adds custom spans on top of auto-instrumentation:

| Span | Purpose |
|------|---------|
| `invoke_workflow` | Root span for the entire workflow execution |
| `execute_node` | Per-node span within the DAG, child of the workflow span |

These custom spans use `@opentelemetry/api` and nest under the auto-instrumented
HTTP spans, giving full trace hierarchy from trigger through node execution to
outbound API calls.

## GenAI Telemetry

The engine includes a GenAI fetch wrapper that detects calls to known LLM API
endpoints (Anthropic, OpenAI) and enriches the auto-created `fetch()` span
with OTel GenAI semantic convention attributes.

**Captured automatically from the API response `usage` object:**

| Attribute | Source |
|-----------|--------|
| `gen_ai.usage.input_tokens` | `response.usage.input_tokens` |
| `gen_ai.usage.output_tokens` | `response.usage.output_tokens` |
| `gen_ai.usage.cache_creation.input_tokens` | `response.usage.cache_creation_input_tokens` |
| `gen_ai.usage.cache_read.input_tokens` | `response.usage.cache_read_input_tokens` |
| `gen_ai.system` | `anthropic` or `openai` (detected from endpoint URL) |
| `gen_ai.request.model` | Model name from request body |
| `gen_ai.response.model` | Model name from response body |
| `gen_ai.response.finish_reasons` | Stop reason from response |
| `gen_ai.operation.name` | `chat`, `embeddings`, etc. |

**NOT captured by default:**
- Prompt content
- Completion content

Content capture is opt-in via the environment variable
`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true`, controlled
per-enclave. Leave it off unless explicitly requested.

## Validating Telemetry Post-Deployment

After deploying a workflow to an observability-enabled cluster:

1. Run the workflow via `wf_run`
2. Wait for completion (check `wf_status`)
3. Verify traces in SigNoz UI -- search by `service.name` matching the workflow name
4. Check that the trace includes:
   - `invoke_workflow` root span
   - `execute_node` child spans for each DAG node
   - `fetch()` spans for outbound HTTP calls
   - GenAI attributes on LLM call spans (if the workflow makes LLM calls)

If traces do not appear:
- Confirm `cluster_profile` shows observability available
- Check that the workflow pod has `OTEL_DENO=true` in its environment
- Check pod logs for OTel-related warnings

## Well-Known DNS Convention

All telemetry routes to a single endpoint:

```
otel-collector.tentacular-observability.svc.cluster.local:4318
```

This is an OTLP/HTTP endpoint (port 4318). The builder injects it as
`OTEL_EXPORTER_OTLP_ENDPOINT` in every workflow pod. The convention holds
regardless of backend -- bundled SigNoz, BYO collector, or external endpoint
all use the same DNS name. Swapping backends is a Service-level change, not a
pod-level change.

## Enclave Scoping

All telemetry carries resource attributes for enclave-scoped filtering:

| Attribute | Source |
|-----------|--------|
| `tentacular.enclave` | Set by `OTEL_RESOURCE_ATTRIBUTES` env var (injected by builder) |
| `k8s.namespace.name` | Set by OTel Collector's `k8s_attributes` processor (cannot be spoofed) |
| `service.name` | Workflow name (from `OTEL_SERVICE_NAME`) |

SigNoz dashboards filter by `tentacular.enclave` to scope views per team.
The Kraken inherently scopes queries by mapping Slack channels to enclaves.

This is **soft tenancy** -- all enclaves share one ClickHouse instance. There
is no hard data isolation between enclaves. Acceptable for single-enterprise
deployments.

## Graceful Degradation

OTel export failures are non-fatal. If the collector is unreachable:

- Deno OTel drops export attempts silently
- The workflow continues executing normally
- No errors, no retries, no backpressure
- The `BasicSink` ring buffer and `/health?detail=1` endpoint remain
  unaffected (they are independent of OTel)

Telemetry is best-effort. Workflow correctness never depends on it.

## Troubleshooting

### OTel silently disabled

Deno's `OTEL_DENO=true` requires `--allow-env` to include the OTel env vars
and `--allow-net` to include the collector endpoint. If either is missing, OTel
is silently disabled -- no error, no warning, just no telemetry.

The builder handles both (`derive.go` adds `OTEL_*` to the allow-env list and
the collector FQDN to allow-net). If you see missing telemetry on a workflow
that predates the observability update, redeploy it to pick up the new
permission flags.

### No GenAI spans

The GenAI wrapper intercepts `globalThis.fetch()`. If a workflow uses a custom
HTTP client that bypasses `fetch()`, GenAI attributes will not be added. Use
standard `fetch()` for LLM API calls.

### Traces appear but with no node spans

The custom `invoke_workflow` and `execute_node` spans require the
`@opentelemetry/api` dependency. Verify it is present in `deno.json` and
resolvable via the module proxy.

### Collector reachable but SigNoz shows no data

Check that the OTel Collector pipeline is configured to export to SigNoz.
This is a Helm chart configuration issue, not a workflow issue. Run the
smoke tests to validate the full pipeline.

## Sensitive Data

The observability system follows defense-in-depth for sensitive data:

1. **Architecture prevents leaks** -- the GenAI wrapper captures metadata
   only (tokens, model, latency), never prompt or completion content
2. **Content capture is opt-in** -- `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`
   defaults to off
3. **Collector redaction** -- the OTel Collector runs a `transform` processor
   that pattern-matches common secret shapes (Bearer tokens, API keys, JWTs)
   and redacts them from span attributes and log bodies
4. **Node code risk** -- `console.log("key: " + secret)` leaks via OTel log
   records, same as any logging system. Do not log secrets.

## BYO Backend

Users can route telemetry to their own observability stack instead of the
bundled SigNoz. The well-known DNS name stays the same -- only the Service
definition changes. See the "Bring Your Own Observability" guide in the
documentation site for step-by-step instructions.
