# Sidecar Containers Reference

Auxiliary containers that run alongside the Deno engine in the same pod,
providing native binary capabilities (ffmpeg, headless browsers, ML models)
via localhost HTTP. Sidecars share a `/shared` emptyDir volume with the
engine and run under the same gVisor sandbox and SecurityContext.

## What Sidecars Are

- Containers declared in `workflow.yaml` that run in the same pod as the engine
- Communicate with the engine via `localhost:PORT` (HTTP or gRPC)
- Share files via `/shared` emptyDir volume (auto-provisioned when sidecars are declared)
- Same pod-level security hardening: gVisor, PSA restricted, `runAsUser: 65534`
- No engine changes needed — nodes call sidecars with `globalThis.fetch()`

## When to Use Sidecars (Decision Rules)

**Use a sidecar when ALL of the following are true:**

1. The workflow needs a native binary that cannot run inside Deno (ffmpeg, Pandoc, ImageMagick, ML models, headless browsers)
2. The tool needs to be called on-demand, potentially multiple times per pod lifetime
3. There is a production-quality Docker image available for the tool (multi-arch preferred)

**Do NOT use a sidecar when:**

- **The task can be done in pure TypeScript/Deno.** If a npm/deno package exists that does the job (image resizing with sharp, PDF generation with jsPDF, data parsing), use it directly in the node. No sidecar needed.
- **The tool is only needed once at startup.** Use an init container instead (tentacular#91) — simpler, no HTTP wrapper needed, no long-running process.
- **The task is a shared heavyweight service** (database, message queue, GPU transcoding pool). Use an exoskeleton service instead — sidecars are per-pod, not shared.
- **You're calling an external API.** Use a contract dependency, not a sidecar. Sidecars are for in-pod native binaries, not for proxying external services.
- **WASM is sufficient.** For lightweight, CPU-cheap operations (hashing, small data transforms), a WASM module may work without the overhead of a second container. But note WASM is 5-10x slower than native under gVisor for CPU-bound work.

### Decision Flowchart

```
Can the task be done in pure TypeScript/Deno?
  YES → Do it in the node. No sidecar.
  NO  ↓
Is it a one-shot preprocessing step (run once, then done)?
  YES → Use an init container (tentacular#91).
  NO  ↓
Is it a shared service (database, queue, inference pool)?
  YES → Use an exoskeleton service.
  NO  ↓
Does a suitable Docker image exist?
  YES → Use a sidecar.
  NO  → Build a custom sidecar image, or reconsider the approach.
```

### Sidecar vs Alternatives

| Approach | When to Use | Performance | Complexity |
|----------|------------|-------------|------------|
| **Pure Deno** | npm/deno package available | Fastest (no IPC) | Lowest |
| **Sidecar** | Native binary, on-demand, per-pod | 5-10x faster than WASM | Medium |
| **Init container** | Native binary, one-shot preprocessing | Same as sidecar | Low |
| **Exoskeleton** | Shared heavyweight service | Varies | High |
| **WASM** | Lightweight native code, no Docker overhead | Slowest under gVisor | Low |
| **Contract dependency** | External API or service | Network-dependent | Lowest |

## Workflow YAML Schema

Sidecars are declared at the top level of `workflow.yaml`, alongside `nodes`
and `edges`:

```yaml
sidecars:
  - name: ffmpeg              # required: [a-z][a-z0-9_-]*, unique per workflow
    image: org/image:tag      # required: container image reference (pin to digest in production)
    port: 9000                # required: 1024-65535, not 8080 (reserved for engine), unique per workflow
    protocol: http            # optional: "http" (default) or "grpc"
    healthPath: /health       # optional: readiness probe path (default: "/health")
    command: []               # optional: override container entrypoint
    args: []                  # optional: override container args
    env:                      # optional: environment variables
      KEY: value
    resources:                # optional: resource requests/limits
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi
```

### Field Rules and Validation

| Field | Required | Constraints | Default |
|-------|----------|-------------|---------|
| `name` | Yes | `[a-z][a-z0-9_-]*`, unique per workflow | — |
| `image` | Yes | Any valid image reference | — |
| `port` | Yes | 1024–65535, not 8080, unique per workflow | — |
| `protocol` | No | `"http"` or `"grpc"` | `"http"` |
| `healthPath` | No | Path string, must return 200 OK | `"/health"` |
| `command` | No | Overrides container ENTRYPOINT | image default |
| `args` | No | Overrides container CMD | image default |
| `env` | No | Map of string key/value pairs | none |
| `resources` | No | Standard Kubernetes resource spec | none |

Port 8080 is reserved for the engine HTTP server. Any sidecar declaring port
8080 will fail validation with a clear error.

## Communication Pattern

Nodes call sidecars via `globalThis.fetch()` — plain HTTP, no special API:

```typescript
import type { Context } from "tentacular";

export default async function run(ctx: Context, input: unknown): Promise<unknown> {
  // Write input file to shared volume
  await Deno.writeFile("/shared/input/video.mp4", inputBytes);

  // Call the sidecar
  const resp = await globalThis.fetch("http://localhost:9000/extract-frames", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      input: "/shared/input/video.mp4",
      fps: 1,
      output_dir: "/shared/output"
    })
  });

  if (!resp.ok) {
    const err = await resp.json();
    throw new Error(`ffmpeg failed: ${err.error}`);
  }

  const result = await resp.json();
  return { frames: result.frames, count: result.count };
}
```

Key rules:
- Use `globalThis.fetch()`, not `ctx.fetch()`. `ctx.fetch()` routes through
  the gateway and is for declared contract dependencies only.
- The builder automatically adds `localhost:PORT` to Deno's `--allow-net` for
  each declared sidecar. No manual flag needed.
- For large files, always use the shared volume (`/shared/input/`, `/shared/output/`)
  rather than HTTP body transfer. Large HTTP bodies cause memory pressure.

### Shared Volume Layout

```
/shared/
  input/     # engine stages input files here
  output/    # sidecar writes output here, engine reads it back
```

Both containers see the same `/shared` emptyDir. Files written by the sidecar
are immediately visible to the engine — no IPC or polling needed.

## Security Model

All sidecar containers inherit the same security hardening as the engine:

| Restriction | Value | Notes |
|-------------|-------|-------|
| `runAsNonRoot` | `true` (pod-level) | All containers run as uid 65534 |
| `runAsUser` | `65534` (pod-level) | Files on `/shared` owned by nobody:nogroup |
| `readOnlyRootFilesystem` | `true` (per container) | Requires `/tmp` emptyDir |
| `allowPrivilegeEscalation` | `false` (per container) | SUID binaries in image are inert |
| `capabilities.drop` | `["ALL"]` (per container) | No Linux capabilities |
| `seccompProfile` | `RuntimeDefault` (pod-level) | gVisor enforces syscall filtering |
| RuntimeClass | `gvisor` | Applies to all containers in the pod |

The builder automatically provisions:
- `/tmp` emptyDir for each sidecar (required for tools that write temp files)
- `/shared` emptyDir for the whole pod (provisioned when any sidecar is declared)

**Network access:** Sidecars can only reach external hosts if the workflow
contract includes a dependency for that host. The builder derives NetworkPolicy
from the contract. A sidecar that needs to download models at startup must have
a corresponding contract dependency.

**Image trust:** There is no curated sidecar registry. Users are responsible
for selecting and pinning images. Pin to a specific digest in production, not
a floating tag.

## Example: ffmpeg Frame Extraction Workflow

```yaml
name: video-frame-extractor
version: "1.0"
description: "Extract frames from video using ffmpeg sidecar"

triggers:
  - type: manual

sidecars:
  - name: ffmpeg
    image: linuxserver/ffmpeg:latest
    port: 9000
    healthPath: /health
    command: ["perl", "-e"]
    args:
      - |
        # Inline HTTP wrapper — see references/sidecar-hooks.md for templates
        use strict; use warnings; use IO::Socket::INET;
        # ... (Perl Tier 2 template) ...
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi

contract:
  version: "1"
  dependencies: {}

nodes:
  fetch-video:
    path: ./nodes/fetch-video.ts
  extract-frames:
    path: ./nodes/extract-frames.ts
  analyze-frames:
    path: ./nodes/analyze-frames.ts

edges:
  - from: fetch-video
    to: extract-frames
  - from: extract-frames
    to: analyze-frames

config:
  timeout: 300s
  retries: 1
```

## Common Sidecar Images

| Image | Use Case | Multi-arch | Hook Required | Available Runtimes |
|-------|----------|-----------|---------------|-------------------|
| `linuxserver/ffmpeg:latest` | Video/audio processing | arm64+amd64 | Yes — no HTTP API | Perl 5.38, bash, nc |
| `pandoc/core:3.6` | Document conversion | arm64+amd64 | No — ships `pandoc-server` | nc only |
| `ghcr.io/browserless/chromium` | Screenshots, PDF generation | arm64+amd64 | No — ships HTTP API | Python3, Node, bash |
| `dpokidov/imagemagick` | Image conversion | TBD | Yes — no HTTP API | Perl, bash |

Use public images with the HTTP hook pattern (`command:`/`args:` injection).
See `references/sidecar-hooks.md` for wrapper templates in Python, Perl,
and bash, and an image compatibility matrix.

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Pod stuck in `Init:0/1` or `PodInitializing` | Sidecar readiness probe failing | Check `healthPath` and port match sidecar's actual health endpoint; check container logs with `wf_logs` |
| `connection refused` on `localhost:PORT` | Sidecar not yet ready when node runs | Add retry logic in the node, or check if the sidecar starts slowly |
| `permission denied` writing to `/shared` | User mismatch between containers | Verify both containers run as uid 65534 — check SecurityContext in generated manifest |
| ffmpeg fails with read-only filesystem error | `/tmp` not mounted | The builder adds `/tmp` emptyDir automatically; if seeing this, check the generated K8s manifest |
| Sidecar needs external network but requests fail | NetworkPolicy blocking outbound | Add a contract dependency for the external host so the builder generates the correct NetworkPolicy egress rule |
| `ctx.fetch() routes to gateway` unexpected | Using `ctx.fetch()` instead of `globalThis.fetch()` | For sidecars, always use `globalThis.fetch("http://localhost:PORT/...")` directly |
| Large HTTP response body causing OOM | Transferring large files in HTTP body | Use `/shared` volume: write file, POST path reference, read output from `/shared/output/` |
| `wf_health` shows sidecar container amber/red | Sidecar crashed or OOM killed | Check `wf_logs` for the sidecar container name; increase memory limit if OOM |

### Checking Sidecar Logs

`wf_logs` returns logs for all containers in the pod. Specify the container
name to narrow output:

```
wf_logs(enclave="my-ns", workflow="video-extractor", container="ffmpeg")
```

The sidecar container name matches the `name` field in the sidecars declaration.
