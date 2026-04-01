# Design: Sidecar Support — tentacular-skill/

## Overview

Add sidecar documentation to the agent skill using the progressive disclosure pattern: a short pointer section in SKILL.md and a full reference file at `references/sidecars.md`.

---

## 1. SKILL.md Section

Insert after the "Scaffold Lifecycle" section (or wherever the workflow spec section ends). The section must be 3-5 lines with a pointer to the reference file.

```markdown
## Sidecars

Workflows can declare sidecar containers for native tools (ffmpeg, headless browsers, ML models). Sidecars run in the same pod, communicate via `localhost:PORT`, and share a `/shared` volume. No engine changes needed — nodes call sidecars with `globalThis.fetch()`.

Read `references/sidecars.md` when:
- Adding native binary capabilities (ffmpeg, Chromium, ImageMagick) to a workflow
- Designing workflows that need shared file handoff between containers
- Debugging sidecar readiness, port conflicts, or SecurityContext issues
```

**Rules followed:**
- Section is exactly 4 lines (1 summary + 3 bullet pointers)
- No implementation details, no YAML examples, no CLI commands inlined
- Pointer uses the standard "Read `references/X.md` when:" format matching existing sections

---

## 2. `references/sidecars.md` Outline

Create new file at `references/sidecars.md`. Structure:

```markdown
# Sidecar Containers

## What Sidecars Are

- Auxiliary containers running alongside the Deno engine in the same pod
- Communicate via localhost HTTP (or gRPC) — no external network needed
- Share files via `/shared` emptyDir volume (automatic when sidecars declared)
- Same security hardening as engine: gVisor, PSA restricted, SecurityContext locked down

## When to Use Sidecars

| Use Case | Example | Why Not WASM/Init |
|----------|---------|-------------------|
| CPU-bound media processing | ffmpeg frame extraction | 5-10x faster than WASM |
| Complex rendering | Headless Chromium screenshots/PDFs | Not WASM-compilable |
| ML inference | Python model serving | Docker image ecosystem |
| Native CLI tools | ImageMagick, Pandoc, wkhtmltopdf | Any Docker image works |

## Workflow YAML Schema

Show the complete `sidecars:` section with all fields, required/optional markers, and defaults.

```yaml
sidecars:
  - name: ffmpeg              # required: [a-z][a-z0-9_-]*, unique per workflow
    image: org/image:tag      # required: container image reference
    command: [...]             # optional: override entrypoint
    args: [...]               # optional: override args
    env:                      # optional: environment variables
      KEY: value
    port: 9000                # required: 1024-65535, not 8080, unique per workflow
    protocol: http            # optional: "http" (default) or "grpc"
    healthPath: /health       # optional: readiness probe path (default: "/health")
    resources:                # optional: resource requests/limits
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi
```

## Communication Pattern

- Nodes call sidecars via `globalThis.fetch("http://localhost:PORT/path")`
- No `ctx.sidecar()` method — plain HTTP fetch, same as any API call
- Deno `--allow-net` automatically includes `localhost:PORT` for each sidecar
- For file-heavy workflows: write to `/shared/input/`, POST to sidecar, read from `/shared/output/`

Example node code:
```typescript
// In a workflow node — call the ffmpeg sidecar
const resp = await globalThis.fetch("http://localhost:9000/extract-frames", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    input: "/shared/input/video.mp4",
    fps: 1,
    output_dir: "/shared/output"
  })
});
const result = await resp.json();
```

## Security Model

- Pod-level: `runAsNonRoot`, `runAsUser: 65534`, `seccompProfile: RuntimeDefault`, gVisor
- Per-container: `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `drop: ALL`
- Network: sidecar external access requires a contract dependency (drives NetworkPolicy)
- Volumes: `/shared` emptyDir (automatic), `/tmp` emptyDir per sidecar (automatic)
- Image trust: user responsibility — no curated registry (yet)

## Common Mistakes

| # | Mistake | What Happens | Fix |
|---|---------|--------------|-----|
| 1 | Using port 8080 for a sidecar | Validation rejects — reserved for engine | Use 9000+ |
| 2 | Forgetting `healthPath` | Defaults to `/health` — sidecar must serve it | Implement GET /health or set custom path |
| 3 | Sidecar needs external access but no contract dep | NetworkPolicy blocks outbound traffic | Add contract dependency for the external host |
| 4 | Using `ctx.fetch()` instead of `globalThis.fetch()` | `ctx.fetch()` routes through the gateway | Use `globalThis.fetch("http://localhost:PORT/...")` |
| 5 | Large files in HTTP body instead of shared volume | Memory pressure, slow transfers | Write to `/shared/input/`, POST path reference |

## Troubleshooting

- **Sidecar not ready**: Check readiness probe path/port match, container logs via `wf_logs`
- **Connection refused on localhost:PORT**: Sidecar may not be ready yet — engine starts before sidecars are ready. Use retry logic or wait for readiness.
- **Permission denied on /shared**: Both containers must run as same uid (65534) — check SecurityContext
- **ffmpeg fails with read-only filesystem**: Sidecar needs `/tmp` emptyDir — builder adds this automatically
```

---

## 3. File Changes Summary

| File | Action |
|------|--------|
| `SKILL.md` | Edit — insert ~5 line sidecar pointer section |
| `references/sidecars.md` | Create — ~120 lines, full sidecar reference |
