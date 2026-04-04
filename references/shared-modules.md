# Shared Node Modules Reference

## Overview

Shared modules are non-DAG TypeScript files in the `nodes/` directory that
are imported by multiple node files. They let you factor out common logic --
auth patterns, formatters, API clients, helpers -- without duplicating code
across nodes.

## How It Works

The builder auto-detects and mounts ALL `.ts` files from `nodes/` into the
workflow pod, not just the files referenced by `workflow.yaml` node entries.
Any `.ts` file in `nodes/` that is not declared as a node in the DAG is
available as an importable module.

Nodes import shared modules using standard relative imports:

```typescript
import { s3Fetch } from "./s3.ts";
```

The import path is relative to `nodes/` because all `.ts` files are mounted
into the same directory inside the pod.

## Example: Shared S3 Auth Module

A workflow that tests RustFS storage and renders a report might share AWS
SigV4 signing logic across two nodes:

```
nodes/
  s3.ts              <-- shared module (not a DAG node)
  test-rustfs.ts     <-- DAG node, imports s3.ts
  render-report.ts   <-- DAG node, imports s3.ts
```

**`nodes/s3.ts`** -- shared signing logic:

```typescript
export async function s3Fetch(
  conn: DependencyConnection,
  method: string,
  path: string,
  body?: BodyInit,
): Promise<Response> {
  const headers = await signV4({
    accessKey: conn.user!,
    secretKey: conn.secret!,
    region: "us-east-1",
    service: "s3",
    method,
    path,
    body,
  });
  return conn.fetch!(path, { method, headers, body });
}

// ... signV4 implementation
```

**`nodes/test-rustfs.ts`** -- uses the shared module:

```typescript
import { s3Fetch } from "./s3.ts";

export default async function run(ctx: Context, input: unknown) {
  const conn = ctx.dependency("tentacular-rustfs");
  const res = await s3Fetch(conn, "PUT", "/test-object", "hello");
  return { status: res.status, ok: res.ok };
}
```

## When to Use Shared Modules

- **Auth patterns:** SigV4 signing, OAuth token refresh, custom header
  construction -- anything that multiple nodes call with different endpoints.
- **Formatters:** Shared output formatting, report generation helpers,
  data transformation utilities.
- **API clients:** Typed wrappers around external APIs that multiple nodes
  consume.
- **Validators:** Input/output validation logic reused across nodes.

General rule: if the same logic appears (or would appear) in 2+ nodes,
extract it into a shared module.

## Anti-Patterns

- **Monolithic utility files.** Do not create a single `utils.ts` that
  accumulates unrelated functions. Keep modules focused by domain: `s3.ts`
  for S3 operations, `format.ts` for output formatting, etc.
- **Shared state.** Modules should be stateless. Do not use module-level
  variables to pass data between nodes -- that is what the DAG edges are for.
- **Re-exporting node functions.** A shared module should contain helper
  logic, not re-export `run()` functions from other nodes. Each node's
  `run()` is its own entry point.

## Naming Conventions

- Name shared modules after their domain: `s3.ts`, `auth.ts`, `format.ts`.
- Do not prefix with `_` or `shared-` -- the builder treats all `.ts` files
  the same regardless of naming.
- Node files declared in `workflow.yaml` should have descriptive names
  matching their DAG role: `test-rustfs.ts`, `render-report.ts`.

## Limitations

- Shared modules must be `.ts` files in the `nodes/` directory. Subdirectories
  inside `nodes/` are not currently supported.
- Shared modules cannot declare their own contract dependencies. Only DAG
  nodes receive `ctx` -- shared modules receive connection objects or data
  passed by the calling node.
- The module proxy resolves third-party imports (e.g., `deno.land/std`).
  Shared modules can use third-party imports just like nodes.
